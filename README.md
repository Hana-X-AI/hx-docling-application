# HX Docling UI

> **Phase 1: Development** | Next.js 16 Web Interface for IBM Granite Docling Document Processing

[![Charter Status](https://img.shields.io/badge/Charter-v0.6.0%20Approved-brightgreen)](project/0.0-charter/0.1-hx-docling-ui-charter.md)
[![Phase](https://img.shields.io/badge/Phase-1%20Development-blue)](project/0.0-charter/0.1-hx-docling-ui-charter.md)
[![License](https://img.shields.io/badge/License-Proprietary-red)]()

**HX Docling UI** is a production-grade web application that provides an intuitive interface for processing documents through IBM's Granite Docling 258M Vision-Language Model. Built on Next.js 16 with TypeScript, it enables users to convert PDFs, Word documents, Excel spreadsheets, PowerPoint presentations, images, and web pages into structured formats (Markdown, HTML, JSON) with real-time progress tracking and persistent history.

**Quick Navigation:** [🚀 Getting Started](#-getting-started) • [🏗️ Architecture](#-architecture) • [📚 Documentation](#-documentation) • [🔄 Development Workflow](#-development-workflow) • [🆘 Support](#-support)

---

## 📑 Table of Contents

- [Features](#-features)
- [Architecture](#-architecture)
- [Technology Stack](#-technology-stack)
- [Getting Started](#-getting-started)
- [Documentation](#-documentation)
- [Project Structure](#-project-structure)
- [Development Workflow](#-development-workflow)
- [Infrastructure](#-infrastructure)
- [Quality Standards](#-quality-standards)
- [Team](#-team)
- [License](#-license)

---

## ✨ Features

### Core Capabilities

- **📄 Multi-Format Document Processing**
  - PDF documents (up to 100 MB)
  - Microsoft Word (.doc, .docx, up to 50 MB)
  - Microsoft Excel (.xls, .xlsx, up to 50 MB)
  - Microsoft PowerPoint (.ppt, .pptx, up to 50 MB)
  - Images (PNG, JPEG, TIFF, up to 25 MB)
  - Web pages via URL

- **🔄 Real-Time Processing**
  - Server-Sent Events (SSE) streaming for live progress updates
  - Automatic reconnection with exponential backoff (up to 30s)
  - Graceful degradation to polling if SSE fails
  - Visual progress indicators across 6 stages

- **💾 Persistent Storage**
  - PostgreSQL-backed job history (90-day retention)
  - Result caching for instant re-download
  - Anonymous session tracking via Redis (24h TTL)
  - File storage with 30-day retention policy

- **📊 Multiple Export Formats**
  - **Markdown**: GitHub Flavored Markdown with YAML frontmatter
  - **HTML**: Standalone with inline Tailwind CSS, dark mode support
  - **JSON**: Full Docling AST hierarchy with metadata
  - **Raw**: Pretty-printed DoclingDocument for debugging

- **🛡️ Enterprise-Grade Reliability**
  - Comprehensive error catalog (E001-E999)
  - Partial result handling (some exports succeed, some fail)
  - MCP error recovery with 3-retry strategy
  - Rate limiting (10 requests/minute per session, HTTP 429 response)
  - Health monitoring for all dependencies

---

## 🏗️ Architecture

### System Overview

```mermaid
flowchart TB
    subgraph Users["👥 Users"]
        U1[Knowledge Worker]
        U2[Data Engineer]
        U3[AI Developer]
    end

    subgraph DevEnv["🖥️ Development Environment<br/>hx-cc-server (192.168.10.224)"]
        subgraph UI["Next.js 16 Application"]
            subgraph Presentation["Presentation Layer"]
                UZ[Upload Zone<br/>Drag & Drop]
                URL[URL Input]
                RV[Results Viewer<br/>Markdown/HTML/JSON]
                HV[History View<br/>Past Jobs]
            end
            
            subgraph Application["Application Layer"]
                ZS[Zustand Store<br/>State Management]
                MC[MCP Client<br/>8 Tools]
                SSE[SSE Manager<br/>Auto-Reconnect]
                EM[Error Manager<br/>E001-E999]
            end
            
            subgraph Data["Data Access Layer"]
                PC[Prisma Client<br/>PostgreSQL]
                RC[Redis Client<br/>Sessions]
                FS[File Storage<br/>/data/docling-uploads/]
            end
        end
    end

    subgraph Infrastructure["🌐 HX-Infrastructure"]
        MCP[hx-docling-mcp-server<br/>192.168.10.217:8000<br/>Document Processing]
        PG[(hx-postgres-server<br/>192.168.10.208:5432<br/>Jobs & Results)]
        RD[(hx-redis-server<br/>192.168.10.209:6379<br/>Sessions)]
        OL3[hx-ollama3-server<br/>192.168.10.206:11434<br/>Granite Docling 258M]
        LL[hx-litellm-server<br/>192.168.10.212:4000<br/>LLM Gateway]
    end

    Users --> DevEnv
    Presentation --> Application
    Application --> Data
    Data --> Infrastructure
    MC -->|HTTP/SSE| MCP
    PC -->|SQL| PG
    RC -->|Redis Protocol| RD
    MCP --> LL
    LL --> OL3

    style Users fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style DevEnv fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style Infrastructure fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Presentation fill:#fff3e0,stroke:#f57c00
    style Application fill:#fce4ec,stroke:#c2185b
    style Data fill:#e0f2f1,stroke:#00796b
```

### Data Flow: Document Processing

```mermaid
sequenceDiagram
    participant User
    participant UI as Next.js UI
    participant Store as Zustand Store
    participant API as API Routes
    participant SSE as SSE Manager
    participant FileStore as File Storage
    participant PG as PostgreSQL
    participant Redis
    participant MCP as MCP Server

    Note over Redis: Session Management
    UI->>Redis: Get/Create Session
    Redis-->>UI: sessionId (24h TTL)

    Note over User,UI: File Upload Flow
    User->>UI: Drop PDF file
    UI->>Store: setFile(file)
    UI->>API: POST /api/upload
    API->>FileStore: Store /data/docling-uploads/{uuid}.pdf
    API->>PG: INSERT Job (status=PENDING)
    PG-->>API: jobId
    API-->>UI: {jobId, fileId}

    Note over User,MCP: Processing Flow
    User->>UI: Click "Process"
    UI->>SSE: Connect /api/process (SSE)
    SSE->>API: POST with jobId
    API->>PG: UPDATE Job (status=PROCESSING)
    API->>MCP: convert_pdf(file)
    
    loop Progress Updates
        MCP-->>API: SSE: {stage, percent, message}
        API-->>SSE: Forward progress
        SSE-->>UI: Update progress bar
        UI->>Store: updateProgress(...)
    end

    alt Network Interruption
        Note over SSE: Connection lost
        SSE->>SSE: Reconnect (1s, 2s, 4s, 8s...)
        SSE->>API: Reconnect with Last-Event-ID
        API-->>SSE: Resume from last state
    end

    MCP-->>API: DoclingDocument
    API->>MCP: export_markdown()
    API->>MCP: export_html()
    API->>MCP: export_json()
    
    alt All Exports Succeed
        API->>PG: INSERT Results (MD, HTML, JSON)
        API->>PG: UPDATE Job (status=COMPLETE)
    else Partial Success
        API->>PG: INSERT Results (available only)
        API->>PG: UPDATE Job (status=PARTIAL_COMPLETE)
    end
    
    API-->>SSE: Complete with results
    SSE-->>UI: Emit complete event
    UI->>Store: setResults({markdown, html, json})
    Store-->>UI: Re-render ResultsViewer
    UI-->>User: Display tabbed results
```

### SSE Resilience Strategy

```mermaid
stateDiagram-v2
    [*] --> Connected: Initial Connection
    
    Connected --> Disconnected: Network Error
    Connected --> Processing: Data Flowing
    Processing --> Disconnected: Connection Lost
    
    Disconnected --> Retry1: Wait 1s
    Retry1 --> Connected: Success
    Retry1 --> Retry2: Fail
    
    Retry2 --> Connected: Wait 2s + Success
    Retry2 --> Retry3: Fail
    
    Retry3 --> Connected: Wait 4s + Success
    Retry3 --> Retry4: Fail
    
    Retry4 --> Connected: Wait 8s + Success
    Retry4 --> Retry5: Fail
    
    Retry5 --> Connected: Wait 16s + Success
    Retry5 --> PollingFallback: Max retries (30s grace)
    
    PollingFallback --> Polling: Every 2s
    Polling --> Connected: SSE Restored
    Polling --> Failed: Timeout
    
    Failed --> [*]: Show Error + Retry Button
    Processing --> Complete: Job Finished
    Complete --> [*]

    note right of Retry1
        Exponential backoff
        Max 30s total grace period
        10 retry attempts
    end note
    
    note right of PollingFallback
        Graceful degradation
        No data loss
        User notified
    end note
```

### Job State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING: Job Created
    
    PENDING --> UPLOADING: File Upload Started
    UPLOADING --> PROCESSING: Upload Complete
    UPLOADING --> ERROR: Upload Failed
    
    PROCESSING --> RETRY_1: MCP Transient Error
    RETRY_1 --> PROCESSING: Retry Success
    RETRY_1 --> RETRY_2: Retry Failed
    
    RETRY_2 --> PROCESSING: Retry Success
    RETRY_2 --> RETRY_3: Retry Failed
    
    RETRY_3 --> PROCESSING: Retry Success
    RETRY_3 --> ERROR: All Retries Exhausted
    
    PROCESSING --> COMPLETE: All Exports Success
    PROCESSING --> PARTIAL_COMPLETE: Some Exports Failed
    PROCESSING --> ERROR: Fatal Error
    
    COMPLETE --> [*]
    PARTIAL_COMPLETE --> [*]
    ERROR --> PENDING: User Retry
    ERROR --> [*]

    note right of RETRY_1
        3-retry strategy
        Exponential backoff
        Preserves partial results
    end note
    
    note right of PARTIAL_COMPLETE
        Available results saved
        User sees which failed
        Can re-attempt failed exports
    end note
```

---

## 🛠️ Technology Stack

### Frontend

| Technology | Version | Purpose |
|------------|---------|---------|
| **Next.js** | 16.x | React framework with App Router |
| **React** | 19.2.x | UI library (ships with Next.js 16) |
| **TypeScript** | 5.1+ | Type safety |
| **Tailwind CSS** | 3.4.x | Utility-first CSS |
| **shadcn/ui** | Latest | Component library |
| **Zustand** | 5.x | State management |
| **Zod** | 3.x | Runtime validation |
| **Lucide React** | Latest | Icon library |

### Backend & Data

| Technology | Version | Purpose |
|------------|---------|---------|
| **Node.js** | 20.9+ | JavaScript runtime |
| **Prisma** | 5.x | PostgreSQL ORM |
| **ioredis** | 5.x | Redis client |
| **PostgreSQL** | Latest | Job & result persistence |
| **Redis** | Latest | Session management |

### AI & Processing

| Technology | Version | Purpose |
|------------|---------|---------|
| **IBM Granite Docling** | 258M | Vision-Language Model |
| **MCP Protocol** | Latest | Model Context Protocol |
| **Ollama** | Latest | Model serving |

---

## 🚀 Getting Started

### Prerequisites

- **Node.js** >= 20.9.0 (LTS)
- **npm** >= 10.0.0
- Access to HX-Infrastructure servers (VPN required)
- PostgreSQL credentials for `docling_db`
- Redis access credentials

### Installation

```bash
# Clone the documentation repository
git clone https://github.com/Hana-X-AI/hx-docling-application.git

# The application will be created separately in Sprint 1.1 at:
# /home/agent0/hx-docling-ui/

# Navigate to application directory (after Sprint 1.1 completion)
cd /home/agent0/hx-docling-ui/

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env.development
# Edit .env.development with your credentials
# For local overrides, also create .env.local (gitignored)

# Generate Prisma client
npx prisma generate

# Run database migrations
npx prisma migrate dev

# Start development server
npm run dev
```

The application will be available at `http://localhost:3000`.

### Environment Variables

Create `.env.development` with the following (and optionally `.env.local` for local overrides):

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

### Verify Installation

```bash
# Type check
npx tsc --noEmit

# Lint
npm run lint

# Run tests
npm run test

# Build
npm run build

# Check infrastructure connectivity
bash scripts/validate-environment.sh
```

---

## 📚 Documentation

### Primary Documents

| Document | Location | Description |
|----------|----------|-------------|
| **Project Charter** | [`project/0.0-charter/0.1-hx-docling-ui-charter.md`](project/0.0-charter/0.1-hx-docling-ui-charter.md) | Complete project specification (v0.6.0, approved) |
| **Charter Reviews** | [`project/0.0-charter/reviews/`](project/0.0-charter/reviews/) | Deep review with 23 findings (all addressed) |
| **Claude Guide** | [`claude.md`](claude.md) | AI assistant development guide (v1.1.0) |
| **Coding Instructions** | [`project/0.6-governance/0.6.0-claude-code-instructions.md`](project/0.6-governance/0.6.0-claude-code-instructions.md) | Detailed technical guidance |
| **Team Roster** | [`project/0.6-governance/0.6.5-team-roster.md`](project/0.6-governance/0.6.5-team-roster.md) | Agent roles and responsibilities |
| **Lessons Learned** | [`project/0.6-governance/0.6.4-lessons-learned.md`](project/0.6-governance/0.6.4-lessons-learned.md) | Past project learnings (22 lessons) |

### Testing Documentation

**Test Framework Overview**:
- **Total Test Cases**: 650 across 6 comprehensive documents
- **Test Documents**: [`project/0.5-test/`](project/0.5-test/)
  - [Master Test Plan Overview](project/0.5-test/00-test-plan-overview.md) - Strategy, scope, quality gates (v2.1.0)
  - [E2E Test Plan](project/0.5-test/01-e2e-test-plan.md) - 91 test cases with Playwright scenarios
  - [Integration Test Plan](project/0.5-test/02-integration-test-plan.md) - 98 test cases for component interaction
  - [API Test Cases](project/0.5-test/03-api-test-cases.md) - 112 test cases for all endpoints
  - [MCP Test Cases](project/0.5-test/04-mcp-test-cases.md) - 142 test cases for protocol compliance
  - [Database Test Cases](project/0.5-test/05-database-test-cases.md) - 102 test cases for PostgreSQL operations
  - [SSE Test Cases](project/0.5-test/06-sse-test-cases.md) - 105 test cases for real-time streaming

**Test Plan Reviews (11-Specialist Consensus)**:
- **Overall Quality Score**: 8.6/10 average (unanimous APPROVED with remediation)
- **Consolidated Assessment**: [`project/0.5-test/reviews/test-plan/consolidated-test-plan-review.md`](project/0.5-test/reviews/test-plan/consolidated-test-plan-review.md) - Complete consensus report
- **Test Plan Technical Review**: [`project/0.5-test/reviews/test-plan/test-plan-review.md`](project/0.5-test/reviews/test-plan/test-plan-review.md) - 9.0/10 score, comprehensive analysis
- **Specialist Reviews** (11 experts): [`project/0.5-test/reviews/test-plan/`](project/0.5-test/reviews/test-plan/)
  - QA Lead (Julia Santos): 8.9/10
  - Next.js Expert (Neo Anderson): 8.2/10 - identifies 45+ Server Component tests needed
  - Pydantic Expert (Paul Warfield): 7.8/10 - identifies 30+ validation tests needed
  - Container Expert (Thomas Docker): 8.6/10 - identifies security scanning as P0
  - API Expert (Bob Martinez): 8.7/10
  - PostgreSQL Expert (Trinity Walsh): 8.9/10
  - Redis Expert (Sri Ramanujan): 8.6/10 - identifies 6+ circuit breaker tests needed
  - FastMCP Expert (George Kim): 8.5/10
  - Docling Expert (Albert Chen): 8.8/10
  - Infrastructure Expert (William Chen): 8.4/10

**Test Data & Dataset Integration**:
- **DocLayNet Dataset**: 80,863 pages, 28GB core + 7.5GB extra data
- **Dataset Analysis Guide**: [`project/0.5-test/test-data/dataset-analysis.md`](project/0.5-test/test-data/dataset-analysis.md)
  - Python data loaders and utilities
  - Test fixture mappings
  - Performance optimization guidance
  - Integration examples

### Quick References

- **Charter Review Summary**: [`project/0.0-charter/reviews/00-start-here.md`](project/0.0-charter/reviews/00-start-here.md)
- **Executive Summary**: [`project/0.0-charter/reviews/executive-summary.md`](project/0.0-charter/reviews/executive-summary.md)
- **Implementation Patterns**: [`project/0.0-charter/reviews/charter-review-implementation-patterns.md`](project/0.0-charter/reviews/charter-review-implementation-patterns.md)
- **Stakeholder Checklist**: [`project/0.0-charter/reviews/charter-review-stakeholder-checklist.md`](project/0.0-charter/reviews/charter-review-stakeholder-checklist.md)

---

## 📁 Project Structure

```
hx-docling-application/
├── project/                              # Project documentation
│   ├── 0.0-charter/                      # Charter v0.6.0 (approved)
│   │   ├── 0.1-hx-docling-ui-charter.md  # Main charter document
│   │   └── reviews/                      # Charter review findings (6 docs)
│   ├── 0.1-plan/                         # Implementation plan
│   │   └── 0.1.1-implementation-plan.md
│   ├── 0.2-architecture/                 # Solution architecture
│   │   ├── 0.2.1-solution-architecture.md
│   │   ├── 0.2.2-agentic-patterns.md
│   │   └── reviews/
│   ├── 0.3-specification/                # Detailed specification
│   │   ├── 0.3.1-detailed-specification.md
│   │   └── reviews/
│   ├── 0.4-tasks/                        # Sprint task definitions
│   │   ├── sprint-0-prerequisites/       # Pre-development setup
│   │   ├── sprint-1.1-scaffold/          # Next.js scaffold
│   │   ├── sprint-1.2-database-redis/    # DB & caching setup
│   │   ├── sprint-1.3-upload/            # File upload component
│   │   ├── sprint-1.4-url-input/         # URL input feature
│   │   ├── sprint-1.5a-mcp-client/       # MCP integration
│   │   ├── sprint-1.5b-sse-progress/     # SSE streaming
│   │   ├── sprint-1.6-results-viewer/    # Results display
│   │   ├── sprint-1.7-history/           # History view
│   │   └── sprint-1.8-testing-docs/      # Testing & documentation
│   ├── 0.5-test/                         # ✨ TEST FRAMEWORK (NEW)
│   │   ├── 00-test-plan-overview.md      # Master test plan (v2.1.0, APPROVED)
│   │   ├── 01-e2e-test-plan.md           # E2E tests (91 tests)
│   │   ├── 02-integration-test-plan.md   # Integration tests (98 tests)
│   │   ├── 03-api-test-cases.md          # API tests (112 tests)
│   │   ├── 04-mcp-test-cases.md          # MCP protocol tests (142 tests)
│   │   ├── 05-database-test-cases.md     # PostgreSQL tests (102 tests)
│   │   ├── 06-sse-test-cases.md          # SSE streaming tests (105 tests)
│   │   ├── A-06-migration-tests-summary.md # Migration test summary
│   │   ├── reviews/                      # 11-specialist consensus reviews
│   │   │   ├── test-plan/                # Comprehensive review documents
│   │   │   │   ├── consolidated-test-plan-review.md (455 lines)
│   │   │   │   ├── test-plan-review.md (1,228 lines, 9.0/10)
│   │   │   │   ├── julia-santos-qa-lead-review.md (970 lines, 8.9/10)
│   │   │   │   ├── neo-anderson-nextjs-review.md (1,116 lines, 8.2/10)
│   │   │   │   ├── paul-warfield-pydantic-review.md (lines, 7.8/10)
│   │   │   │   ├── thomas-docker-containers-review.md (583 lines, 8.6/10)
│   │   │   │   ├── bob-martinez-api-review.md (486 lines, 8.7/10)
│   │   │   │   ├── trinity-walsh-postgresql-review.md (404 lines, 8.9/10)
│   │   │   │   ├── sri-ramanujan-redis-review.md (565 lines, 8.6/10)
│   │   │   │   ├── george-kim-fastmcp-review.md (351 lines, 8.5/10)
│   │   │   │   ├── albert-chen-docling-review.md (758 lines, 8.8/10)
│   │   │   │   └── william-chen-infrastructure-review.md (404 lines, 8.4/10)
│   │   │   ├── e2e/ ├── integration/
│   │   │   ├── api/ ├── mcp/
│   │   │   ├── database/ ├── sse/
│   │   │   └── test-data/                # Test data reviews
│   │   └── test-data/                    # Test datasets & fixtures
│   │       ├── dataset-analysis.md       # DocLayNet integration guide
│   │       ├── test-fixtures/            # Fixture definitions
│   │       ├── mock-responses/           # Mock response factories
│   │       └── [DocLayNet files organized by type]
│   ├── 0.6-governance/                   # Governance documents
│   │   ├── 0.6.0-claude-code-instructions.md
│   │   ├── 0.6.1-raidd-log.md
│   │   ├── 0.6.2-status-report.md
│   │   ├── 0.6.3-backlog.md
│   │   ├── 0.6.4-lessons-learned.md
│   │   ├── 0.6.5-team-roster.md
│   │   ├── 0.6.6-defect-log.md
│   │   └── reviews/
│   ├── 0.7-templates/                    # Document templates
│   │   ├── charter-template.md
│   │   ├── plan-template.md
│   │   ├── spec-template.md
│   │   └── tasks-template.md
│   └── 0.8-references/                   # Reference materials
│       ├── agentic-design-patterns-docs-main/
│       ├── DocLayNet/                    # Dataset documentation
│       ├── granite-docling-258M-WebGPU/  # Model reference
│       ├── constitution.md
│       ├── docling-mcp-service-descriptor.md
│       ├── hx-shadcn-service-descriptor.md
│       ├── service-descriptor.md
│       └── task-024-completion-summary.md
├── claude.md                             # AI assistant guide (v1.1.0)
├── README.md                             # This file
├── .gitignore                            # Git ignore rules
└── x-docs/                               # Archived documentation

# Application structure (to be created in Sprint 1.1 at /home/agent0/hx-docling-ui/)
hx-docling-ui/
├── src/
│   ├── app/                              # Next.js App Router
│   │   ├── layout.tsx                    # Root layout
│   │   ├── page.tsx                      # Main page
│   │   ├── history/page.tsx              # History view
│   │   └── api/                          # API routes
│   │       ├── upload/route.ts           # File upload
│   │       ├── process/route.ts          # SSE processing
│   │       ├── history/route.ts          # Job history
│   │       └── health/route.ts           # Health check
│   ├── components/
│   │   ├── ui/                           # shadcn/ui components
│   │   ├── upload/                       # Upload components
│   │   ├── processing/                   # Progress components
│   │   ├── results/                      # Result viewers
│   │   └── history/                      # History components
│   ├── lib/
│   │   ├── db/prisma.ts                  # Prisma client
│   │   ├── redis/client.ts               # Redis client
│   │   ├── mcp/                          # MCP integration (8 tools)
│   │   ├── sse/                          # SSE manager + reconnection
│   │   └── validation/                   # Zod schemas
│   ├── stores/                           # Zustand stores
│   ├── hooks/                            # Custom React hooks
│   └── types/                            # TypeScript types
├── prisma/
│   ├── schema.prisma                     # Database schema
│   └── migrations/                       # Migration files
├── scripts/
│   ├── cleanup-uploads.sh                # File cleanup cron
│   └── validate-environment.sh           # Infrastructure check
├── tests/                                # Test suites
├── .env.local                            # Environment variables (gitignored)
├── next.config.ts                        # Next.js config
├── tailwind.config.ts                    # Tailwind config
├── tsconfig.json                         # TypeScript config
└── package.json                          # Dependencies
```

---

## 🔄 Development Workflow

### Sprint Sequence (Phase 1: 9-10 Sessions)

```mermaid
gantt
    title Phase 1 Development Timeline
    dateFormat  YYYY-MM-DD
    section Setup
    Sprint 1.1 - Scaffold          :s1, 2025-12-12, 1d
    Sprint 1.2 - Database          :s2, after s1, 1d
    section Core Features
    Sprint 1.3 - Upload            :s3, after s2, 1d
    Sprint 1.4 - URL Input         :s4, after s3, 1d
    Sprint 1.5 - MCP Client        :s5, after s4, 1d
    section Checkpoint
    Integration Checkpoint         :milestone, after s5, 0d
    section Results & History
    Sprint 1.6 - Results Viewer    :s6, after s5, 1d
    Sprint 1.7 - History View      :s7, after s6, 1d
    section Polish
    Sprint 1.8 - Testing & Docs    :s8, after s7, 1d
    Phase 1 Complete               :milestone, after s8, 0d
```

| Sprint | Focus | Deliverables | Duration |
|--------|-------|--------------|----------|
| **1.1** | Project Scaffold | Next.js 16 setup, shadcn/ui, Prisma schema | 1 session |
| **1.2** | Database Setup | PostgreSQL config, Redis client, migrations | 1 session |
| **1.3** | Upload Component | Drag-drop, validation, file storage | 1 session |
| **1.4** | URL Input | URL validation, submission | 1 session |
| **1.5** | MCP Integration | 8 tools, SSE streaming, error handling | 1 session |
| **🔍 CHECKPOINT** | **Validation** | **MCP health, DB tests, E2E smoke test** | **Mandatory** |
| **1.6** | Results Viewer | Markdown/HTML/JSON tabs, download | 1 session |
| **1.7** | History View | Past jobs, pagination, re-download | 1 session |
| **1.8** | Polish & Testing | Unit tests, E2E tests, documentation | 1-2 sessions |

### Quality Gates

Every commit must pass:

```bash
# 1. Type checking
npx tsc --noEmit

# 2. Linting
npm run lint

# 3. Unit tests
npm run test

# 4. Build verification
npm run build

# 5. Prisma validation
npx prisma validate
```

### Code Standards

- **TypeScript**: Zero errors, strict mode enabled
- **ESLint**: Zero warnings/errors
- **Test Coverage**: 100% for utils/libs, 80%+ for components
- **Accessibility**: Lighthouse score >= 90
- **Performance**: Lighthouse score >= 80

---

## 🌐 Infrastructure

### HX-Infrastructure Servers

| Server | IP | Port | Purpose | Status |
|--------|-----|------|---------|--------|
| **hx-cc-server** | 192.168.10.224 | 3000 | Development environment | 🟡 Setup in Sprint 1.1 |
| **hx-docling-mcp-server** | 192.168.10.217 | 8000 | Document processing (MCP) | ✅ Operational |
| **hx-postgres-server** | 192.168.10.208 | 5432 | Job & result persistence | 🟡 Setup in Sprint 1.2 |
| **hx-redis-server** | 192.168.10.209 | 6379 | Session tracking | 🟡 Setup in Sprint 1.2 |
| **hx-ollama3-server** | 192.168.10.206 | 11434 | Granite Docling 258M model | ✅ Operational |
| **hx-litellm-server** | 192.168.10.212 | 4000 | LLM gateway | ✅ Operational |

### MCP Tools (Phase 1: 8 Tools)

| Tool | File Types | Purpose | Status |
|------|------------|---------|--------|
| `convert_pdf` | .pdf | PDF conversion | ✅ Available |
| `convert_docx` | .doc, .docx | Word conversion | ✅ Available |
| `convert_xlsx` | .xls, .xlsx | Excel conversion | ✅ Available |
| `convert_pptx` | .ppt, .pptx | PowerPoint conversion | ✅ Available |
| `convert_url` | URLs | Web page conversion | ✅ Available |
| `export_markdown` | - | Export to Markdown | ✅ Available |
| `export_html` | - | Export to HTML | ✅ Available |
| `export_json` | - | Export to JSON | ✅ Available |

### Backlog Tools (Phase 2+)

13 additional MCP tools documented in charter Section 18: `generate_title`, `generate_toc`, `generate_sections`, `generate_headings`, `generate_paragraphs`, `generate_lists`, `generate_tables`, `generate_images`, `generate_code_blocks`, `generate_references`, `generate_knowledge_graph`, `split_document`, `merge_documents`.

---

## ✅ Quality Standards

### Acceptance Criteria (20 Total)

Phase 1 complete when all criteria met:

- [ ] Application runs at `http://hx-cc-server.hx.dev.local:3000`
- [ ] User can drag-drop PDF and see Markdown output
- [ ] User can drag-drop Word (.docx) and see output
- [ ] User can drag-drop Excel (.xlsx) and see output
- [ ] User can drag-drop PowerPoint (.pptx) and see output
- [ ] User can paste URL and see HTML output
- [ ] Progress stages display during processing
- [ ] SSE reconnects automatically after network drop
- [ ] Results viewer shows Markdown/HTML/JSON tabs
- [ ] Download functionality works for all formats
- [ ] Processing results persist to PostgreSQL
- [ ] Job history displays past conversions
- [ ] User can re-download past results
- [ ] Session tracking works via Redis
- [ ] Error codes display with recovery actions
- [ ] Zero TypeScript errors
- [ ] Zero ESLint errors
- [ ] All unit tests pass (100% coverage for utils)
- [ ] `npm run build` succeeds
- [ ] No console errors in browser

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Largest Contentful Paint (LCP) | < 2.5s | Lighthouse |
| First Contentful Paint (FCP) | < 1.8s | Lighthouse |
| Time to first progress update | < 500ms | Manual timing |
| SSE reconnection time | < 30s | Network simulation |
| Accessibility score | >= 90 | Lighthouse |
| Performance score | >= 80 | Lighthouse |

### Error Code Reference

<details>
<summary>📋 Quick Reference for Error Codes (E001-E999)</summary>

| Range | Category | Example Codes | Description |
|-------|----------|---------------|-------------|
| **E0xx** | File Errors | E001, E002, E003 | File too large, invalid type, corrupt file |
| **E1xx** | URL Errors | E101, E102, E103 | Invalid URL format, unreachable, SSRF blocked |
| **E2xx** | MCP Errors | E201, E202, E203 | MCP unavailable, timeout, invalid response |
| **E3xx** | Processing Errors | E301, E302, E303 | Processing timeout, partial failure, quota exceeded |
| **E4xx** | Database Errors | E401, E402, E403 | Connection failed, query error, persist failed |
| **E5xx** | Session Errors | E501, E502, E503 | Session expired, invalid session, Redis unavailable |
| **E6xx** | Rate Limiting | E601, E602 | Rate limit exceeded, quota exhausted |
| **E9xx** | Network Errors | E901, E902, E903 | SSE disconnected, reconnect failed, network timeout |

**Full error catalog**: See [`claude.md` Section 9](claude.md#9-troubleshooting) for detailed recovery actions.

</details>

---

## 🧪 Test Framework & Quality Assurance

### Test Coverage Summary (v2.1.0)

**Overall Status**: ✅ **APPROVED** (11-specialist consensus review, average 8.6/10)

| Test Category | Count | Document | Status | Quality Score |
|---------------|-------|----------|--------|---------|
| **End-to-End (E2E)** | 91 | [01-e2e-test-plan.md](project/0.5-test/01-e2e-test-plan.md) | ✅ Complete | Vitest + Playwright |
| **Integration** | 98 | [02-integration-test-plan.md](project/0.5-test/02-integration-test-plan.md) | ✅ Complete | Component + API integration |
| **API Endpoints** | 112 | [03-api-test-cases.md](project/0.5-test/03-api-test-cases.md) | ✅ Complete | 8.7/10 |
| **MCP Protocol** | 142 | [04-mcp-test-cases.md](project/0.5-test/04-mcp-test-cases.md) | ✅ Complete | JSON-RPC 2.0 compliance |
| **PostgreSQL** | 102 | [05-database-test-cases.md](project/0.5-test/05-database-test-cases.md) | ✅ Complete | 8.9/10 |
| **SSE Streaming** | 105 | [06-sse-test-cases.md](project/0.5-test/06-sse-test-cases.md) | ✅ Complete | Real-time progress tracking |
| **TOTAL** | **650** | [Master Plan](project/0.5-test/00-test-plan-overview.md) | **✅ COMPREHENSIVE** | **9.0/10** |

### 11-Specialist Consensus Review

**Consolidated Verdict**: APPROVED FOR IMPLEMENTATION with mandatory remediation

**Expert Panel** (11 specialist reviews totaling ~8,600 lines):

| Expert | Focus Area | Score | Status | Key Finding |
|--------|-----------|-------|--------|------------|
| Julia Santos | QA Strategy & Gates | 8.9/10 | ✅ APPROVED | Excellent test foundation |
| Neo Anderson | Next.js Framework | 8.2/10 | ⚠️ CONDITIONAL | 45+ Server Component tests needed |
| Paul Warfield | Pydantic Validation | 7.8/10 | ⚠️ CONDITIONAL | 30+ validation tests needed |
| Thomas Docker | Container Security | 8.6/10 | ⚠️ CONDITIONAL | Security scanning required (P0) |
| Bob Martinez | API Testing | 8.7/10 | ✅ APPROVED | 112 endpoint tests sufficient |
| Trinity Walsh | PostgreSQL Operations | 8.9/10 | ✅ APPROVED | 102 database tests excellent |
| Sri Ramanujan | Redis/Cache Patterns | 8.6/10 | ⚠️ CONDITIONAL | 6+ circuit breaker tests needed |
| George Kim | FastMCP Gateway | 8.5/10 | ✅ APPROVED | Protocol compliance strong |
| Albert Chen | Docling Architecture | 8.8/10 | ✅ APPROVED | MCP integration comprehensive |
| William Chen | Infrastructure/Ops | 8.4/10 | ⚠️ CONDITIONAL | Load testing expansion needed |
| **AI Assessment System** | **Overall Technical** | **9.0/10** | **✅ APPROVED** | **650 tests well-distributed** |
| | | **AVG: 8.6/10** | **11/11 CONSENSUS** | |

📄 **Full Review**: [consolidated-test-plan-review.md](project/0.5-test/reviews/test-plan/consolidated-test-plan-review.md)

### Test Framework Architecture

```mermaid
flowchart TB
    subgraph Tests["650 Comprehensive Test Cases"]
        E2E["E2E Tests (91)"]
        Integration["Integration Tests (98)"]
        API["API Tests (112)"]
        MCP["MCP Tests (142)"]
        DB["Database Tests (102)"]
        SSE["SSE Tests (105)"]
    end
    
    subgraph Tools["Testing Tools & Frameworks"]
        Vitest["Vitest - Unit/Component<br/>Fast in-memory test runner"]
        Playwright["Playwright - E2E<br/>Cross-browser automation"]
        MSW["MSW - API Mocking<br/>Network request interception"]
        Locust["Locust - Performance<br/>Load & stress testing"]
    end
    
    subgraph Quality["Quality Gates"]
        TS["TypeScript<br/>Zero Errors"]
        Lint["ESLint<br/>Zero Warnings"]
        Coverage["Test Coverage<br/>100% utils, 80%+ components"]
        A11y["Accessibility<br/>Lighthouse >= 90"]
        Perf["Performance<br/>Lighthouse >= 80"]
    end
    
    Tests --> Tools
    Tests --> Quality
    
    style Tests fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Tools fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Quality fill:#fff3e0,stroke:#f57c00,stroke-width:2px
```

### Critical Remediation Roadmap

**Consensus Gap Analysis** (8+ specialists identified these):

#### Phase 1: Release Blockers (MUST FIX - Week 1-2)

| Gap | Priority | Tests Needed | Owner | Effort |
|-----|----------|------------|-------|--------|
| **Next.js Server Components** | P0 | 45+ | Neo Anderson | 40-50 hrs |
| **Pydantic Validation** | P0 | 30+ | Paul Warfield | 30-40 hrs |
| **Container Security Scanning** | P0 | 10+ | Thomas Docker | 15-20 hrs |
| **Redis Circuit Breaker** | P0 | 6+ | Sri Ramanujan | 10-15 hrs |

**Phase 1 Total**: 80-100 hours, **BLOCKING FOR GA**

#### Phase 2: High Priority (Week 3-4)

| Gap | Priority | Tests Needed | Owner | Effort |
|-----|----------|------------|-------|--------|
| **Load Testing Expansion** | P1 | 10+ | Bob Martinez, William Chen | 20-25 hrs |
| **Database Migration** | P1 | 6+ | Trinity Walsh | 10-12 hrs |
| **Error Handling** | P1 | 4+ | George Kim | 8-10 hrs |
| **API Concurrency** | P1 | 5+ | Bob Martinez | 8-10 hrs |

**Phase 2 Total**: 40-50 hours, **HIGH: Should complete before GA**

#### Phase 3: Medium Priority (Sprint 2)

| Gap | Priority | Tests Needed | Owner | Effort |
|-----|----------|------------|-------|--------|
| **Disaster Recovery** | P2 | 5+ | William Chen | 10-15 hrs |
| **Chaos Engineering** | P2 | 8+ | William Chen | 15-20 hrs |
| **Backup/Recovery** | P2 | 6+ | Trinity Walsh | 10-12 hrs |

**Phase 3 Total**: 30-40 hours

### Dataset Integration: DocLayNet

**Status**: ✅ Integrated with local deployment ready

- **Dataset Size**: 80,863 pages total
  - Core: 28GB (30,230 PNG images + COCO JSON annotations)
  - Extra: 7.5GB (50,633 PDF documents + token JSON files)
- **Integration Guide**: [dataset-analysis.md](project/0.5-test/test-data/dataset-analysis.md)
- **Python Utilities**: Data loaders, test fixtures, performance helpers
- **Test Coverage**: Complete fixture mappings for all 650 tests
- **Location**: `/home/agent0/hx-docling-application/project/0.5-test/test-data/`

**Available Test Fixtures**:
- PDF documents (multiple sizes and complexity levels)
- Word documents (.docx, various structures)
- Excel spreadsheets (single & multi-sheet)
- PowerPoint presentations (mixed content types)
- Web pages (HTML snapshots)
- Image files (PNG, JPEG, TIFF)
- Edge cases (corrupted files, oversized documents, invalid formats)

### Test Metrics & Quality Gates

**Master Test Plan Quality**: 9.0/10

**Blocker Status**: ✅ **ALL RESOLVED**
- ✅ Test data specification (DocLayNet fully integrated)
- ✅ SSRF prevention (security tested in API suite)
- ✅ Performance SLAs (defined and measured)

**Release Readiness**: 🟡 **CONDITIONAL**
- ✅ Core testing foundation solid (650 tests)
- ✅ API/Database/SSE/MCP coverage comprehensive
- ⚠️ Framework-specific gaps require Phase 1 work
- ⚠️ Must address P0 items before production

**Next Steps**:
1. Review [consolidated assessment](project/0.5-test/reviews/test-plan/consolidated-test-plan-review.md)
2. Prioritize Phase 1 remediation with team
3. Allocate 80-100 hours for critical gap closure
4. Plan implementation timeline with development sprints

---

### Project Leadership

| Role | Name | Responsibility |
|------|------|----------------|
| **CAIO** | Jarvis Richardson (Agent Zero) | Strategic oversight, final approval |
| **Orchestration** | Agent Zero | Multi-agent coordination, quality gates |

### Core Team (Phase 1)

| Role | Agent | Focus Area |
|------|-------|------------|
| **Platform Architect** | Alex Rivera | ADR, architecture validation |
| **Testing & QA** | Julia Santos | Test strategy, quality validation |
| **Database & Infra** | William Chen | PostgreSQL, Redis, file storage |
| **Primary Developer** | Trinity | Next.js implementation |
| **Frontend UI** | Ola Mae Johnson | UI/UX, accessibility |
| **MCP Integration** | James Dean | MCP client library (8 tools) |

### Phase 2 Team (Deferred)

| Role | Agent | Focus Area |
|------|-------|------------|
| **DNS & Certificates** | Frank Lucas | SSL, reverse proxy |
| **Docker & Compose** | Thomas Anderson | Containerization, deployment |

---

## 🎯 Project Status

```mermaid
gantt
    title Project Milestones
    dateFormat  YYYY-MM-DD
    section Planning
    Charter Creation           :done, 2025-12-01, 5d
    Charter Review             :done, 2025-12-06, 3d
    Charter Approved (v0.6.0)  :milestone, done, 2025-12-11, 0d
    section Documentation
    Test Plan v2.1.0           :done, 2025-12-12, 3d
    11-Specialist Reviews      :done, 2025-12-13, 2d
    Test Framework Complete    :milestone, done, 2025-12-15, 0d
    section Phase 1
    Sprint 1.1-1.2 (Setup)     :crit, active, 2025-12-12, 2d
    Sprint 1.3-1.5 (Core)      :2025-12-14, 3d
    Integration Checkpoint     :milestone, 2025-12-17, 0d
    Sprint 1.6-1.8 (Polish)    :2025-12-17, 3d
    Phase 1 Complete           :milestone, 2025-12-20, 0d
    section Phase 2
    Phase 2 Charter            :2025-12-21, 3d
    Deployment                 :2025-12-24, 5d
```

| Milestone | Status | Date | Details |
|-----------|--------|------|---------|
| Charter v0.6.0 Approved | ✅ Complete | 2025-12-11 | All 23 findings addressed |
| All 23 Review Findings Addressed | ✅ Complete | 2025-12-11 | Charter review consensus |
| Team Roster Approved | ✅ Complete | 2025-12-11 | 6 core agents assigned |
| Coding Instructions Complete | ✅ Complete | 2025-12-11 | Technical guidance finalized |
| Claude Guide v1.1.0 | ✅ Complete | 2025-12-11 | AI assistant manual |
| **Test Framework v2.1.0** | **✅ Complete** | **2025-12-15** | **650 tests, 9.0/10 score** |
| **11-Specialist Reviews** | **✅ Complete** | **2025-12-15** | **8.6/10 avg, consensus APPROVED** |
| **DocLayNet Integration** | **✅ Complete** | **2025-12-15** | **80,863 pages locally deployed** |
| **Phase 1 Development** | 🟡 In Progress | 2025-12-12+ | Sprint 1.1-1.8 execution |
| Phase 2 Deployment | ⏸️ Planned | 2025-12-21+ | Post-Phase 1 launch |

### Recent Git History

- **Commit 0443ad3** (2025-12-15): Comprehensive test documentation, dataset analysis, 11-specialist reviews, remediation roadmap
  - 13 files changed
  - 51,657 insertions
  - Test framework (650 tests) fully documented
  - DocLayNet integration guide added
  - Phase 1-3 remediation roadmap defined

---

## 🔐 Security & Compliance

- **Authentication**: Anonymous sessions (AD integration deferred to Phase 2)
- **Rate Limiting**: 10 requests/minute per session
- **Input Validation**: Zod schemas for all user input
- **File Upload Security**: Type validation, size limits, MIME type checks
- **URL Security**: SSRF prevention, allowlist domains
- **Data Retention**: 
  - Files: 30 days
  - Job history: 90 days
  - Sessions: 24 hours
- **Error Handling**: No sensitive data in error messages
- **Logging**: Structured logging with sanitized PII

---

## 📄 License

**Proprietary** - Copyright © 2025 Hana-X-AI. All rights reserved.

This software is proprietary and confidential. Unauthorized copying, distribution, or use is strictly prohibited.

---

## 🤝 Contributing

### Phase 1 Development Process

1. **Review Documentation**
   - Read [`claude.md`](claude.md) for AI assistant guidance
   - Review charter Section 16.2 for sprint details
   - Check [`project/0.6-governance/0.6.4-lessons-learned.md`](project/0.6-governance/0.6.4-lessons-learned.md)

2. **Development Workflow**
   ```bash
   # Create feature branch
   git checkout -b sprint-1.x/feature-name
   
   # Make changes
   # Run quality gates
   npm run lint && npm run test && npm run build
   
   # Commit with conventional commits
   git commit -m "feat(component): add upload zone"
   
   # Push and create PR
   git push origin sprint-1.x/feature-name
   ```

3. **Code Review Requirements**
   - All quality gates pass (see Section 5.3 in `claude.md`)
   - TypeScript/ESLint zero errors
   - Test coverage maintained
   - Accessibility score >= 90
   - Architecture review by Alex Rivera
   - Testing sign-off by Julia Santos

---

## 🆘 Support

### Troubleshooting

See [`claude.md` Section 9](claude.md#9-troubleshooting) for common issues:
- TypeScript/Prisma errors
- Database connectivity
- Redis connection failures
- MCP server errors
- File upload issues
- SSE disconnections
- Rate limiting

### Infrastructure Health Check

```bash
# Run validation script
bash scripts/validate-environment.sh

# Manual checks
curl http://hx-docling-mcp-server.hx.dev.local:8000/health
redis-cli -h hx-redis-server.hx.dev.local ping
pg_isready -h hx-postgres-server.hx.dev.local
```

**Expected Health Response:**

```json
{
  "status": "healthy",
  "timestamp": "2025-12-11T14:30:00Z",
  "version": "1.0.0",
  "checks": {
    "mcp": { 
      "status": "ok", 
      "latency_ms": 45,
      "url": "http://hx-docling-mcp-server.hx.dev.local:8000"
    },
    "postgres": { 
      "status": "ok", 
      "latency_ms": 12,
      "connection": "hx-postgres-server.hx.dev.local:5432"
    },
    "redis": { 
      "status": "ok", 
      "latency_ms": 3,
      "connection": "hx-redis-server.hx.dev.local:6379"
    }
  }
}
```

### Contact

- **Project Lead**: Agent Zero (@agent-zero)
- **Technical Questions**: Alex Rivera (@alex)
- **Testing Questions**: Julia Santos (@julia)
- **Infrastructure**: William Chen (@william)

---

## 🎉 Acknowledgments

- **IBM Research** for Granite Docling 258M VLM
- **Docling Team** for MCP server implementation
- **HX-Infrastructure Team** for platform support
- **Code Review Team** for comprehensive charter review (23 findings, all addressed)

---

<div align="center">

**Built with ❤️ by the HX-Infrastructure Team**

[![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.1+-blue?logo=typescript)](https://www.typescriptlang.org/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind-3.4+-38B2AC?logo=tailwind-css)](https://tailwindcss.com/)
[![Prisma](https://img.shields.io/badge/Prisma-5-2D3748?logo=prisma)](https://www.prisma.io/)

[Documentation](project/0.0-charter/0.1-hx-docling-ui-charter.md) • [Charter Reviews](project/0.0-charter/reviews/) • [Claude Guide](claude.md) • [Team Roster](project/0.6-governance/0.6.5-team-roster.md)

</div>
